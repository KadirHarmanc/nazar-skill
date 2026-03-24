# iOS App Store Submission Rules

22 rules that Apple reviews during App Store submission. Each rule includes what to check, where to check, pass/fail conditions, and fix guidance.

---

## Rule 1: Info.plist Permission Descriptions

- **Rule Name**: Privacy Usage Descriptions
- **What File to Check**: `Info.plist` (or `**/Info.plist` in subdirectories)
- **What to Look For**: Every `NS*UsageDescription` key must have a non-empty, user-facing string value. Common keys:
  - `NSCameraUsageDescription`
  - `NSPhotoLibraryUsageDescription`
  - `NSPhotoLibraryAddUsageDescription`
  - `NSMicrophoneUsageDescription`
  - `NSLocationWhenInUseUsageDescription`
  - `NSLocationAlwaysUsageDescription`
  - `NSLocationAlwaysAndWhenInUseUsageDescription`
  - `NSContactsUsageDescription`
  - `NSCalendarsUsageDescription`
  - `NSBluetoothAlwaysUsageDescription`
  - `NSFaceIDUsageDescription`
  - `NSMotionUsageDescription`
  - `NSSpeechRecognitionUsageDescription`
  - `NSHealthShareUsageDescription`
  - `NSHealthUpdateUsageDescription`
- **Pass Condition**: All usage description keys present in Info.plist have a meaningful, non-empty string (not placeholder text). Also verify that any framework usage in code that requires a permission has the matching key in Info.plist.
- **Fail Reason**: Missing or empty usage description causes immediate rejection. Apple requires a clear explanation of why the app needs each permission.
- **Fix Guide**: Add a descriptive, user-friendly string for each `NS*UsageDescription` key. Example: `"NSCameraUsageDescription": "We need camera access to let you take profile photos."` Grep the codebase for `AVCaptureDevice`, `CLLocationManager`, `CNContactStore`, etc. to find which permissions are actually used.

---

## Rule 2: PrivacyInfo.xcprivacy Exists

- **Rule Name**: Privacy Manifest File
- **What File to Check**: `PrivacyInfo.xcprivacy` (or `**/PrivacyInfo.xcprivacy`)
- **What to Look For**: The file must exist in the project root or app target directory.
- **Pass Condition**: File exists and is valid XML/plist format.
- **Fail Reason**: Starting iOS 17 / Spring 2024, Apple requires a privacy manifest for all apps and SDKs. Missing this file will cause rejection.
- **Fix Guide**: Create `PrivacyInfo.xcprivacy` in the app target. Use Xcode > File > New > App Privacy. See `privacy-manifest.md` for required contents.

---

## Rule 3: Required Reason API Declarations

- **Rule Name**: Required Reason APIs
- **What File to Check**: `PrivacyInfo.xcprivacy` and source files (`*.swift`, `*.m`, `*.mm`)
- **What to Look For**: If the app uses any Required Reason APIs, they must be declared in `NSPrivacyAccessedAPITypes` within `PrivacyInfo.xcprivacy`. Required Reason APIs include:
  - File timestamp APIs (`NSFileCreationDate`, `NSFileModificationDate`, `stat`, `getattrlist`)
  - Disk space APIs (`NSFileSystemFreeSize`, `NSFileSystemSize`, `statfs`)
  - User defaults APIs (`UserDefaults`)
  - System boot time APIs (`systemUptime`, `mach_absolute_time`)
  - Active keyboard APIs
- **Pass Condition**: Every Required Reason API used in code is declared with the correct reason code in `PrivacyInfo.xcprivacy`.
- **Fail Reason**: Using a Required Reason API without declaring it in the privacy manifest causes rejection.
- **Fix Guide**: Add each API type and its reason code to `NSPrivacyAccessedAPITypes` in `PrivacyInfo.xcprivacy`. Refer to Apple's documentation for valid reason codes.

---

## Rule 4: No UIWebView Usage

- **Rule Name**: UIWebView Deprecation
- **What File to Check**: All source files (`*.swift`, `*.m`, `*.mm`, `*.h`), `*.xib`, `*.storyboard`, and `Podfile.lock` / `Package.resolved`
- **What to Look For**: Any reference to `UIWebView`.
- **Pass Condition**: Zero references to `UIWebView` in the entire project (including dependencies).
- **Fail Reason**: Apple deprecated UIWebView and rejects apps containing it. This includes transitive usage via third-party libraries.
- **Fix Guide**: Replace `UIWebView` with `WKWebView`. Check third-party SDKs by running `grep -r "UIWebView" Pods/` or checking binary frameworks. Update outdated dependencies that still use UIWebView.

---

## Rule 5: App Transport Security (ATS)

- **Rule Name**: ATS Configuration
- **What File to Check**: `Info.plist`
- **What to Look For**: `NSAppTransportSecurity` dictionary. Check for:
  - `NSAllowsArbitraryLoads` = `true` (blanket exception - BAD)
  - `NSAllowsArbitraryLoadsInWebContent` (acceptable for web browsers)
  - `NSExceptionDomains` with specific domains (acceptable with justification)
- **Pass Condition**: No `NSAllowsArbitraryLoads = true` blanket exception. Specific domain exceptions are acceptable.
- **Fail Reason**: Blanket ATS exceptions signal poor security and cause rejection or require justification that is usually denied.
- **Fix Guide**: Remove `NSAllowsArbitraryLoads`. Use HTTPS for all endpoints. If specific domains require HTTP, add them as `NSExceptionDomains` with justification.

---

## Rule 6: App Icons - All Required Sizes

- **Rule Name**: App Icon Completeness
- **What File to Check**: `Assets.xcassets/AppIcon.appiconset/Contents.json` (or `**/AppIcon.appiconset/Contents.json`)
- **What to Look For**: Required icon sizes:
  - 1024x1024 (App Store)
  - 180x180 (iPhone @3x)
  - 120x120 (iPhone @2x)
  - 167x167 (iPad Pro @2x)
  - 152x152 (iPad @2x)
  - 76x76 (iPad @1x)
  - 40x40, 60x60, 58x58, 80x80, 87x87, 120x120 (Spotlight, Settings)
- **Pass Condition**: `Contents.json` references image files for all required sizes and the image files actually exist.
- **Fail Reason**: Missing app icon sizes cause build validation errors and submission rejection.
- **Fix Guide**: Generate all required icon sizes from a 1024x1024 source image. Use Xcode asset catalog or a tool like `appicon.co`.

---

## Rule 7: Launch Screen

- **Rule Name**: Launch Screen Exists
- **What File to Check**: `LaunchScreen.storyboard` or `LaunchScreen.xib` or Info.plist key `UILaunchStoryboardName`
- **What to Look For**: A launch screen storyboard file exists and is referenced in Info.plist or the project target.
- **Pass Condition**: `LaunchScreen.storyboard` (or equivalent) exists and `UILaunchStoryboardName` is set in Info.plist.
- **Fail Reason**: Apps without a launch screen are rejected. Launch images (static PNG approach) are deprecated since iOS 13.
- **Fix Guide**: Create `LaunchScreen.storyboard` with appropriate branding. Set `UILaunchStoryboardName` to `"LaunchScreen"` in Info.plist.

---

## Rule 8: Bundle Identifier Format

- **Rule Name**: Valid Bundle Identifier
- **What File to Check**: `Info.plist` (`CFBundleIdentifier`), `*.pbxproj`
- **What to Look For**: `CFBundleIdentifier` must be a valid reverse domain format (e.g., `com.company.appname`). Must not contain wildcards or placeholder values.
- **Pass Condition**: Bundle identifier matches pattern `^[a-zA-Z][a-zA-Z0-9-]*(\.[a-zA-Z][a-zA-Z0-9-]*)+$` and does not contain `$(PRODUCT_BUNDLE_IDENTIFIER)` unresolved or placeholder text.
- **Fail Reason**: Invalid bundle identifier prevents submission entirely.
- **Fix Guide**: Set a proper reverse-domain bundle identifier in the project settings. Example: `com.yourcompany.yourapp`.

---

## Rule 9: Minimum iOS Version

- **Rule Name**: Minimum Deployment Target
- **What File to Check**: `*.pbxproj` (`IPHONEOS_DEPLOYMENT_TARGET`), `Podfile` (`platform :ios`)
- **What to Look For**: The minimum deployment target version.
- **Pass Condition**: Minimum iOS version is 16.0 or higher for new app submissions (as of 2024+).
- **Fail Reason**: Apple requires new apps to support at least the latest major iOS version minus two. Targeting below iOS 16 may cause rejection for new submissions.
- **Fix Guide**: Update `IPHONEOS_DEPLOYMENT_TARGET` to `16.0` or higher in build settings. Update `Podfile` platform accordingly.

---

## Rule 10: Push Notification Entitlements

- **Rule Name**: Push Notification Configuration
- **What File to Check**: `*.entitlements` files, `Info.plist`, source code for push registration
- **What to Look For**: If the app uses push notifications (`UNUserNotificationCenter`, `registerForRemoteNotifications`), the `aps-environment` entitlement must be present.
- **Pass Condition**: If push is used in code, `aps-environment` entitlement exists. If push is not used, this rule passes automatically.
- **Fail Reason**: Registering for push without the proper entitlement causes crashes and rejection.
- **Fix Guide**: Enable Push Notifications capability in Xcode. Ensure `*.entitlements` contains `aps-environment` key set to `development` or `production`.

---

## Rule 11: In-App Purchase Configuration

- **Rule Name**: StoreKit / IAP Setup
- **What File to Check**: Source files for `SKProduct`, `SKPaymentQueue`, `Product`, `StoreKit`, `*.storekit` configuration files
- **What to Look For**: If the app uses in-app purchases, StoreKit must be properly configured.
- **Pass Condition**: If IAP code exists, StoreKit entitlement is present and products are configured. If no IAP code, passes automatically.
- **Fail Reason**: IAP code without proper configuration causes rejection. Also, offering purchases outside StoreKit violates guidelines.
- **Fix Guide**: Add StoreKit capability in Xcode. Create product configurations in App Store Connect. Use `StoreKit 2` API for new development.

---

## Rule 12: IPv6 Compatibility

- **Rule Name**: IPv6 Network Compatibility
- **What File to Check**: All source files (`*.swift`, `*.m`, `*.mm`)
- **What to Look For**: Hardcoded IPv4 addresses (pattern: `\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}`). Check for `inet_addr`, `inet_aton`, `sockaddr_in` (not `sockaddr_in6`).
- **Pass Condition**: No hardcoded IPv4 addresses. Uses hostname-based networking or supports dual-stack.
- **Fail Reason**: Apple tests on IPv6-only networks. Hardcoded IPv4 addresses cause the app to fail on these networks, leading to rejection.
- **Fix Guide**: Replace hardcoded IP addresses with hostnames. Use `Network.framework` or `URLSession` which handle IPv6 automatically. Never use raw IPv4 socket connections.

---

## Rule 13: 64-bit Architecture Requirement

- **Rule Name**: 64-bit Architecture Support
- **What File to Check**: `*.pbxproj` (`ARCHS`, `VALID_ARCHS`), build settings
- **What to Look For**: Architecture settings must include `arm64`. Must not be limited to `armv7` or `i386` only.
- **Pass Condition**: `ARCHS` includes `arm64`. No `armv7`-only configurations.
- **Fail Reason**: 32-bit only apps have been rejected since iOS 11. All apps must support 64-bit.
- **Fix Guide**: Set `ARCHS` to `$(ARCHS_STANDARD)` which includes `arm64`. Remove any explicit `armv7` or `i386` overrides.

---

## Rule 14: IDFA / ATT Compliance

- **Rule Name**: App Tracking Transparency
- **What File to Check**: Source files, `Info.plist`, framework imports
- **What to Look For**: If `AdSupport.framework` or `ASIdentifierManager` or `advertisingIdentifier` is used, `ATTrackingManager` must also be implemented and `NSUserTrackingUsageDescription` must be in Info.plist.
- **Pass Condition**: If IDFA is accessed, ATT prompt is implemented and usage description exists. If IDFA is not used, passes automatically.
- **Fail Reason**: Accessing IDFA without ATT prompt is a guaranteed rejection since iOS 14.5.
- **Fix Guide**: Import `AppTrackingTransparency`. Call `ATTrackingManager.requestTrackingAuthorization` before accessing IDFA. Add `NSUserTrackingUsageDescription` to Info.plist.

---

## Rule 15: Third-Party SDK Privacy Manifests

- **Rule Name**: SDK Privacy Manifests
- **What File to Check**: `Pods/` directory, `*.xcframework` bundles, `Package.resolved`
- **What to Look For**: Third-party SDKs listed on Apple's required list must include their own `PrivacyInfo.xcprivacy`. Key SDKs requiring manifests: Alamofire, Firebase, Facebook SDK, Google Analytics, Amplitude, Mixpanel, OneSignal, Sentry, Adjust, AppsFlyer.
- **Pass Condition**: All third-party SDKs that require privacy manifests have them bundled. Check inside `.xcframework` or `Pods/[SDK]/PrivacyInfo.xcprivacy`.
- **Fail Reason**: Starting Spring 2024, apps including SDKs on Apple's list without privacy manifests are rejected.
- **Fix Guide**: Update all third-party SDKs to their latest versions that include privacy manifests. Check each SDK's release notes for privacy manifest support.

---

## Rule 16: No Debug/Development Mode Indicators

- **Rule Name**: Release Build Configuration
- **What File to Check**: Source files, `*.pbxproj`, `Info.plist`, `*.xcconfig`
- **What to Look For**:
  - `#if DEBUG` blocks that should not ship (e.g., test servers, debug menus)
  - `CONFIGURATION = Debug` in archive scheme
  - `GCC_PREPROCESSOR_DEFINITIONS` containing `DEBUG=1` in Release config
  - Hardcoded staging/development API URLs
  - Test/debug UI elements left in
- **Pass Condition**: Release configuration does not contain debug flags. No development URLs in release code paths.
- **Fail Reason**: Apps shipped with debug indicators may expose sensitive data or test functionality, leading to rejection.
- **Fix Guide**: Verify build scheme uses Release configuration for Archive. Ensure `#if DEBUG` properly guards all test code. Use build configurations to switch API endpoints.

---

## Rule 17: No Placeholder/Lorem Ipsum Content

- **Rule Name**: Production Content Check
- **What File to Check**: All source files, `*.storyboard`, `*.xib`, `*.strings`, asset files
- **What to Look For**: Patterns: `lorem ipsum`, `placeholder`, `TODO`, `FIXME`, `test@test.com`, `example.com` (in user-facing strings), `John Doe`, `Jane Doe`, `sample text`, `dummy data`.
- **Pass Condition**: No placeholder or lorem ipsum text found in user-facing strings or UI files.
- **Fail Reason**: Placeholder content signals an incomplete app and causes rejection.
- **Fix Guide**: Search the entire project for placeholder patterns and replace with real content. Check storyboards and XIBs for placeholder label text.

---

## Rule 18: Crash Risk Patterns - Force Unwrap

- **Rule Name**: Force Unwrap Safety
- **What File to Check**: All `*.swift` files
- **What to Look For**: Excessive use of force unwrap (`!`) on optionals, especially:
  - `as!` force casts
  - `try!` force try
  - `!` on variables that could be nil (e.g., `dictionary["key"]!`)
  - `implicitlyUnwrappedOptional` in function parameters
- **Pass Condition**: Fewer than 10 force unwraps in the codebase (excluding IBOutlets which are standard). No force unwraps on network response data or user input.
- **Fail Reason**: Force unwraps on uncertain data cause crashes. Frequent crashes lead to rejection or removal from the store.
- **Fix Guide**: Replace `!` with `guard let` / `if let` optional binding. Use `as?` instead of `as!`. Use `try?` or `do-catch` instead of `try!`. Keep `!` only for IBOutlets and genuinely guaranteed non-nil values.

---

## Rule 19: Background Modes Validation

- **Rule Name**: Background Modes Usage
- **What File to Check**: `Info.plist` (`UIBackgroundModes`), source code
- **What to Look For**: Each declared background mode must be actually used in code:
  - `audio` - must use `AVAudioSession`
  - `location` - must use `CLLocationManager` with `allowsBackgroundLocationUpdates`
  - `fetch` - must use `BGAppRefreshTask`
  - `remote-notification` - must handle silent push
  - `processing` - must use `BGProcessingTask`
  - `bluetooth-central` or `bluetooth-peripheral` - must use `CoreBluetooth`
- **Pass Condition**: Every declared background mode has corresponding code usage. No unused background modes declared.
- **Fail Reason**: Declaring background modes without using them is a rejection reason. Apple sees it as an attempt to keep the app alive unnecessarily.
- **Fix Guide**: Remove any background mode from `UIBackgroundModes` that is not actually used in code. Only declare modes that the app genuinely needs.

---

## Rule 20: App Extension Bundle Configuration

- **Rule Name**: App Extension Setup
- **What File to Check**: Extension targets in `*.pbxproj`, extension `Info.plist` files
- **What to Look For**: If app extensions exist (widgets, share extensions, etc.):
  - Extension bundle ID must be prefixed with main app bundle ID
  - Extension has its own `Info.plist` with `NSExtension` dictionary
  - Extension `NSExtensionPointIdentifier` is valid
  - Extension minimum deployment target matches or exceeds main app
- **Pass Condition**: All extensions are properly configured with correct bundle ID hierarchy and valid extension point identifiers. If no extensions, passes automatically.
- **Fail Reason**: Misconfigured extensions cause build validation failure and rejection.
- **Fix Guide**: Ensure extension bundle ID follows format `com.company.app.extensionname`. Verify each extension's `Info.plist` has proper `NSExtension` configuration.

---

## Rule 21: Keychain Sharing Entitlements

- **Rule Name**: Keychain Configuration
- **What File to Check**: `*.entitlements` files
- **What to Look For**: If `keychain-access-groups` is declared, the values must use the correct App ID prefix format: `$(AppIdentifierPrefix)com.company.app`.
- **Pass Condition**: Keychain access groups use valid format with `$(AppIdentifierPrefix)` or correct team ID prefix. If no keychain sharing, passes automatically.
- **Fail Reason**: Incorrect keychain access group format causes signing issues and submission failure.
- **Fix Guide**: Use `$(AppIdentifierPrefix)` prefix for keychain access groups. Enable Keychain Sharing capability in Xcode to auto-configure.

---

## Rule 22: Associated Domains Configuration

- **Rule Name**: Associated Domains
- **What File to Check**: `*.entitlements` files, source code for universal links
- **What to Look For**: If `com.apple.developer.associated-domains` entitlement is present:
  - Domains must use `applinks:`, `webcredentials:`, or `activitycontinuation:` prefix
  - Domain format must be valid (e.g., `applinks:example.com`)
  - No wildcard subdomains (e.g., `applinks:*.example.com` is invalid for applinks)
  - The server must host `.well-known/apple-app-site-association` (cannot be verified locally but flag if entitlement exists)
- **Pass Condition**: Associated domains use valid format and prefixes. If not using associated domains, passes automatically.
- **Fail Reason**: Malformed associated domains cause universal links to fail silently, which can lead to poor review experience and rejection.
- **Fix Guide**: Use correct format: `applinks:yourdomain.com`. Ensure your server hosts the AASA file at `https://yourdomain.com/.well-known/apple-app-site-association`. Do not use wildcards with `applinks:`.
