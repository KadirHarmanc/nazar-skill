# Mobile Security Rules

Security patterns for React Native, Flutter, and Expo mobile applications.

---

# React Native (10 Patterns)

## 1. AsyncStorage with Secrets

- **Name:** rn-asyncstorage-secrets
- **Regex:** `(AsyncStorage\.(setItem|multiSet)\s*\(\s*["\'].*(token|secret|password|key|credential|auth)["\'])`
- **Severity:** critical
- **Description:** Detects sensitive data being stored in AsyncStorage, which is unencrypted plaintext storage on the device. Attackers with physical or root access can read all values.
- **FP Indicators:**
  - Storing non-sensitive keys like user preferences or UI state
  - The value is encrypted before storage
  - Located in test/mock files
- **Fix:** Use `expo-secure-store` (Expo) or `react-native-keychain` for sensitive data. AsyncStorage should only hold non-sensitive preferences.

---

## 2. Console Log Sensitive Data

- **Name:** rn-console-sensitive
- **Regex:** `(console\.(log|debug|info|warn)\s*\(.*\b(token|password|secret|auth|credential|session|jwt|bearer)\b)`
- **Severity:** high
- **Description:** Detects console logging of sensitive data in React Native, which appears in device logs accessible via `adb logcat` or Xcode console.
- **FP Indicators:**
  - Logging the word but not the value (e.g., `"token refreshed"`)
  - Inside `__DEV__` conditional blocks
  - Test files with mock data
- **Fix:** Remove all sensitive data logging. Use `__DEV__` guards for debug logs. Use a logging library with production log level filtering.

---

## 3. Deep Link Validation Missing

- **Name:** rn-deeplink-validation
- **Regex:** `(Linking\.(addEventListener|getInitialURL)|useURL\s*\(|DeepLinking\.(addRoute|addScheme)|navigation\.navigate\s*\(.*url)`
- **Severity:** high
- **Description:** Detects deep link handlers that may not validate incoming URLs, potentially allowing malicious apps to trigger unintended navigation or actions.
- **FP Indicators:**
  - URL is validated/parsed before navigation
  - A whitelist of allowed routes is checked
  - Located in test code
- **Fix:** Always validate and sanitize deep link URLs before processing. Maintain a whitelist of allowed deep link routes. Never pass deep link parameters directly to sensitive operations.

---

## 4. Insecure Network Config

- **Name:** rn-insecure-network
- **Regex:** `(NSAppTransportSecurity.*NSAllowsArbitraryLoads\s*.*true|android:usesCleartextTraffic\s*=\s*["\']true|cleartextTrafficPermitted.*true)`
- **Severity:** high
- **Description:** Detects insecure network configurations that allow cleartext HTTP traffic, disabling OS-level transport security protections.
- **FP Indicators:**
  - Development-only configuration (debug build variant)
  - Scoped to specific localhost domains only
  - Located in test configuration
- **Fix:** Remove `NSAllowsArbitraryLoads` and `usesCleartextTraffic=true`. Add specific domain exceptions only for domains that truly require HTTP.

---

## 5. Hardcoded API Endpoints

- **Name:** rn-hardcoded-endpoints
- **Regex:** `(fetch\s*\(\s*["\']https?://[a-zA-Z0-9]|axios\.\w+\s*\(\s*["\']https?://[a-zA-Z0-9]|baseURL\s*[:=]\s*["\']https?://[a-zA-Z0-9])`
- **Severity:** medium
- **Description:** Detects hardcoded API base URLs that make environment switching difficult and may leak staging/internal endpoints in production builds.
- **FP Indicators:**
  - Points to a well-known public API (e.g., Google, AWS, CDN)
  - Located in environment config files
  - Test files with mock server URLs
- **Fix:** Use environment variables or a config module to define API endpoints. Use build-time injection for environment-specific URLs.

---

## 6. Bridge Security - Sensitive Native Modules

- **Name:** rn-bridge-security
- **Regex:** `(NativeModules\.\w+\.(getToken|getSecret|getPassword|decrypt|getKey)|@ReactMethod.*\b(secret|password|token|key)\b|RCT_EXPORT_METHOD.*\b(secret|password|token|key)\b)`
- **Severity:** high
- **Description:** Detects native bridge methods that expose sensitive operations to the JavaScript layer, which can be intercepted or manipulated.
- **FP Indicators:**
  - Method is protected by authentication checks
  - The native module uses secure enclave/keychain
  - Located in test harness code
- **Fix:** Minimize sensitive data crossing the JS bridge. Perform sensitive operations entirely in native code. Use secure storage APIs on the native side and only return operation results (not raw secrets).

---

## 7. Insecure WebView Configuration

- **Name:** rn-webview-insecure
- **Regex:** `(javaScriptEnabled\s*[=:]\s*\{?\s*true|injectedJavaScript\s*=|onMessage\s*=.*postMessage|allowFileAccess\s*[=:]\s*\{?\s*true|mixedContentMode\s*=\s*["\']always)`
- **Severity:** high
- **Description:** Detects insecure WebView configurations that may enable XSS, arbitrary file access, or insecure content mixing.
- **FP Indicators:**
  - WebView loads only trusted first-party content
  - JavaScript injection is from a trusted static source
  - Located in development/debug screens
- **Fix:** Restrict WebView to HTTPS origins. Validate all `postMessage` data. Disable file access. Set `mixedContentMode` to `"never"`. Use `originWhitelist` to restrict allowed domains.

---

## 8. Sensitive Data in Redux/State

- **Name:** rn-state-sensitive
- **Regex:** `(redux-persist.*\b(token|password|secret|key)\b|persistReducer.*\b(auth|session|credential)\b|MMKV\.(set|getString)\s*\(.*\b(token|password|secret)\b)`
- **Severity:** high
- **Description:** Detects sensitive data persisted in state management stores (Redux, MMKV) which may use unencrypted storage.
- **FP Indicators:**
  - State is encrypted before persistence
  - Only stores non-sensitive auth state (e.g., `isLoggedIn: true`)
  - MMKV is configured with encryption
- **Fix:** Do not persist sensitive tokens in Redux state. Use encrypted storage (react-native-keychain, expo-secure-store). If using MMKV, enable encryption mode.

---

## 9. Certificate Pinning Missing

- **Name:** rn-cert-pinning
- **Regex:** `(TrustKit|ssl-pinning|react-native-ssl-pinning|certificatePinning|pinning)`
- **Severity:** medium
- **Description:** Checks for the presence (or absence) of certificate pinning configuration. Lack of pinning allows MITM attacks with rogue certificates.
- **FP Indicators:**
  - This is a presence check - finding the pattern is good
  - Absence of this pattern in production network code is the actual concern
  - Development/staging configs intentionally skip pinning
- **Fix:** Implement certificate pinning using `react-native-ssl-pinning` or TrustKit. Pin to the leaf or intermediate certificate. Implement pin rotation strategy.

---

## 10. Biometric Auth Bypass

- **Name:** rn-biometric-bypass
- **Regex:** `(LocalAuthentication|TouchID|FaceID|react-native-biometrics|authenticateAsync)\s*.*\.(then|catch)\s*\(\s*\(\s*\)\s*=>.*navigate`
- **Severity:** high
- **Description:** Detects biometric authentication that may be bypassable - e.g., navigating to protected screens in catch blocks or without verifying the auth result.
- **FP Indicators:**
  - Catch block shows an error and does not navigate
  - Auth result is properly checked before navigation
  - Located in test code
- **Fix:** Always verify the authentication result object. Never navigate to protected screens in error/catch handlers. Use biometric auth in combination with server-side session validation.

---

# Flutter (10 Patterns)

## 11. SharedPreferences Secrets

- **Name:** flutter-sharedprefs-secrets
- **Regex:** `(SharedPreferences.*\.(setString|setInt)\s*\(\s*["\'].*(token|secret|password|key|credential|auth)["\']|prefs\.(setString|set)\s*\(\s*["\'].*(token|password|secret))`
- **Severity:** critical
- **Description:** Detects sensitive data stored in SharedPreferences, which is plaintext XML/plist storage accessible on rooted/jailbroken devices.
- **FP Indicators:**
  - Storing non-sensitive preferences (theme, locale)
  - Value is encrypted before storage
  - Test/mock code
- **Fix:** Use `flutter_secure_storage` for sensitive data. SharedPreferences should only store non-sensitive user preferences.

---

## 12. DevTools Debug Mode

- **Name:** flutter-devtools-debug
- **Regex:** `(debugShowCheckedModeBanner\s*:\s*true|assert\s*\(\s*debugPaintSizeEnabled|debugPrint\s*\(|kDebugMode\s*(?!=)(?!.*if))`
- **Severity:** medium
- **Description:** Detects debug mode indicators and debug utilities that should not be present in release builds.
- **FP Indicators:**
  - Inside `assert()` statements (stripped in release)
  - Gated behind `kDebugMode` or `kReleaseMode` checks
  - Development-only debug screens
- **Fix:** Use `kDebugMode` or `kReleaseMode` guards for all debug code. Set `debugShowCheckedModeBanner: false` in release. Use `dart:developer` log with level filtering.

---

## 13. Platform Channel Validation

- **Name:** flutter-platform-channel
- **Regex:** `(MethodChannel\s*\(\s*["\']|EventChannel\s*\(\s*["\']|BasicMessageChannel\s*\()`
- **Severity:** medium
- **Description:** Detects platform channel usage that may lack proper input validation or error handling when communicating between Dart and native code.
- **FP Indicators:**
  - Channel has input validation on both sides
  - Used for simple non-sensitive operations (device info, permissions)
  - Well-known plugin channels (e.g., Flutter official plugins)
- **Fix:** Validate all data crossing platform channels on both Dart and native sides. Handle `PlatformException` properly. Do not pass sensitive data through channels without encryption.

---

## 14. Insecure HTTP Client

- **Name:** flutter-insecure-http
- **Regex:** `(HttpClient\s*\(\)\.badCertificateCallback\s*=|SecurityContext.*allowLegacyUnsafeRenegotiation|http://[a-zA-Z0-9].*(?!localhost|127\.0\.0\.1))`
- **Severity:** high
- **Description:** Detects insecure HTTP client configurations that bypass certificate validation or allow cleartext traffic.
- **FP Indicators:**
  - Development-only configuration
  - `badCertificateCallback` is used for certificate pinning (returns false for invalid certs)
  - Localhost/emulator URLs
- **Fix:** Never set `badCertificateCallback` to always return true. Use proper certificate validation. Ensure all production traffic uses HTTPS.

---

## 15. Obfuscation Missing

- **Name:** flutter-obfuscation
- **Regex:** `(flutter\s+build\s+(apk|appbundle|ipa|ios)(?!.*--obfuscate))`
- **Severity:** medium
- **Description:** Detects Flutter build commands that do not include the `--obfuscate` flag, making reverse engineering of the compiled Dart code easier.
- **FP Indicators:**
  - Debug/development builds
  - CI/CD pipeline that adds flags separately
  - Build scripts that compose the command dynamically
- **Fix:** Always use `flutter build apk --obfuscate --split-debug-info=build/debug-info` for release builds. Add obfuscation to CI/CD release pipeline.

---

## 16. Hardcoded Firebase Config

- **Name:** flutter-firebase-config
- **Regex:** `(apiKey\s*:\s*["\']AIza[a-zA-Z0-9_-]{35}|google-services\.json|GoogleService-Info\.plist|FirebaseOptions\s*\(.*apiKey\s*:\s*["\'])`
- **Severity:** medium
- **Description:** Detects hardcoded Firebase configuration that may include API keys directly in source code rather than in properly managed config files.
- **FP Indicators:**
  - Firebase API keys are inherently public (client-side)
  - Config is loaded from environment/build config
  - Located in generated config files (firebase_options.dart)
- **Fix:** Use `firebase_core` with `DefaultFirebaseOptions` from auto-generated config. Ensure Firestore/RTDB security rules are properly configured regardless of key exposure.

---

## 17. Dio Interceptor Logging

- **Name:** flutter-dio-logging
- **Regex:** `(LogInterceptor\s*\(|dio\.interceptors\.add\s*\(\s*LogInterceptor|requestBody\s*:\s*true|responseBody\s*:\s*true)`
- **Severity:** medium
- **Description:** Detects Dio HTTP client log interceptors that may dump full request/response bodies including auth headers and sensitive data.
- **FP Indicators:**
  - Gated behind `kDebugMode`
  - Custom interceptor that redacts sensitive headers
  - Test-only Dio instances
- **Fix:** Remove `LogInterceptor` from production builds. Use conditional interceptor registration: `if (kDebugMode) dio.interceptors.add(LogInterceptor())`. Redact Authorization headers.

---

## 18. Insecure Local Database

- **Name:** flutter-insecure-db
- **Regex:** `(openDatabase\s*\((?!.*password)|Hive\.openBox\s*\((?!.*encryptionCipher)|objectbox.*Store\s*\((?!.*encrypt))`
- **Severity:** high
- **Description:** Detects local databases opened without encryption, leaving stored data readable on rooted/jailbroken devices.
- **FP Indicators:**
  - Database stores only non-sensitive data (cached UI, app state)
  - Encryption is configured elsewhere in the codebase
  - Test/development database
- **Fix:** Use `sqflite_sqlcipher` for encrypted SQLite. Use `Hive` with `HiveAesCipher`. Enable encryption on ObjectBox. For sensitive data, consider using the platform keychain.

---

## 19. Screenshot Prevention Missing

- **Name:** flutter-screenshot-prevention
- **Regex:** `(FLAG_SECURE|SecureApplication|secure\s*:\s*true|WindowManager\.LayoutParams\.FLAG_SECURE)`
- **Severity:** low
- **Description:** Checks for screenshot/screen recording prevention on sensitive screens. Absence may allow sensitive data capture.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - Not all screens need screenshot prevention
  - Only critical for financial/health/auth screens
- **Fix:** Apply `FLAG_SECURE` on Android and equivalent on iOS for screens displaying sensitive data. Use `secure_application` Flutter package.

---

## 20. Root/Jailbreak Detection Missing

- **Name:** flutter-root-detection
- **Regex:** `(flutter_jailbreak_detection|root_checker|freeRASP|isJailBroken|isRooted|SafetyNet|DeviceIntegrity)`
- **Severity:** medium
- **Description:** Checks for root/jailbreak detection implementation. Absence means the app runs without device integrity verification.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - Low-risk apps may not need root detection
  - Detection is handled by an MDM/EMM solution
- **Fix:** Implement root/jailbreak detection using `flutter_jailbreak_detection` or `freeRASP`. Decide on behavior: warn user, restrict features, or block access based on risk level.

---

# Expo (5 Patterns)

## 21. Expo SecureStore Not Used

- **Name:** expo-securestore-missing
- **Regex:** `(AsyncStorage\.(setItem|getItem|multiSet)\s*\(.*\b(token|secret|password|auth|session|jwt)\b)`
- **Severity:** critical
- **Description:** Detects sensitive data stored in AsyncStorage instead of expo-secure-store in Expo projects. AsyncStorage is unencrypted.
- **FP Indicators:**
  - Non-sensitive data stored in AsyncStorage
  - The value is encrypted before storage
  - Expo SecureStore is used elsewhere for the actual secrets
- **Fix:** Replace AsyncStorage with `expo-secure-store` for all sensitive data: `await SecureStore.setItemAsync(key, value)`. AsyncStorage is fine for non-sensitive preferences.

---

## 22. Expo OTA Update Security

- **Name:** expo-ota-security
- **Regex:** `(expo-updates|Updates\.(checkForUpdateAsync|fetchUpdateAsync)|runtimeVersion|"updates"\s*:\s*\{)`
- **Severity:** medium
- **Description:** Detects OTA update configuration that should be reviewed for security. Unsigned or unvalidated updates can be a vector for code injection.
- **FP Indicators:**
  - Updates are configured with proper code signing
  - Using EAS Update with built-in integrity checks
  - Development/preview builds
- **Fix:** Enable code signing for OTA updates. Use `runtimeVersion` policy to prevent incompatible updates. Configure update URL to use HTTPS only. Use EAS Update for managed integrity verification.

---

## 23. Expo Permissions Over-Request

- **Name:** expo-permissions-overrequest
- **Regex:** `(expo-camera|expo-location|expo-contacts|expo-calendar|expo-media-library|expo-sensors|expo-audio).*import`
- **Severity:** low
- **Description:** Flags permission-requiring Expo modules for review. Over-requesting permissions degrades user trust and may trigger app store review flags.
- **FP Indicators:**
  - Each permission is actually used in the app
  - Permissions are requested contextually (not all at startup)
  - Located in feature modules that genuinely need the permission
- **Fix:** Audit all permission-requiring modules. Remove unused ones. Request permissions contextually when the feature is first used, not at app launch. Explain to users why each permission is needed.

---

## 24. Expo Dev Client in Production

- **Name:** expo-dev-client-prod
- **Regex:** `(expo-dev-client|expo-dev-launcher|expo-dev-menu|devClient\s*:\s*true|"developmentClient"\s*:\s*true)`
- **Severity:** high
- **Description:** Detects Expo development client packages or configuration that should not be included in production builds.
- **FP Indicators:**
  - Listed in devDependencies only
  - Gated behind development environment checks
  - Located in development configuration files
- **Fix:** Ensure `expo-dev-client` is only in `devDependencies`. Verify production builds exclude development tools. Use separate app.config.js profiles for development and production.

---

## 25. Expo Config Secrets

- **Name:** expo-config-secrets
- **Regex:** `(app\.(json|config\.(js|ts))).*\b(apiKey|secret|password|token|AKIA|sk-|ghp_)\b|extra\s*:\s*\{[^}]*(apiKey|secret|password|token)`
- **Severity:** critical
- **Description:** Detects secrets hardcoded in Expo app configuration files (app.json, app.config.js) which are bundled into the app binary.
- **FP Indicators:**
  - Values read from environment variables at build time
  - Public client IDs that are not sensitive
  - Placeholder/example values in template configs
- **Fix:** Use environment variables in `app.config.js`: `extra: { apiKey: process.env.API_KEY }`. Never hardcode secrets in app.json. Use EAS Secrets for build-time injection.
