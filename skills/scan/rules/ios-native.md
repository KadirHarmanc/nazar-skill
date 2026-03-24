# iOS Native Security Rules

Security patterns for Swift and Objective-C iOS applications.

---

## 1. NSUserDefaults Secrets

- **Name:** ios-nsuserdefaults-secrets
- **Regex:** `(UserDefaults\.(standard\.set|standard\.string|standard\.object)\s*\(.*\b(token|secret|password|key|credential|auth|session|jwt)\b|NSUserDefaults.*\b(token|secret|password|key|credential)\b|\[\s*NSUserDefaults\s+standardUserDefaults\s*\].*\b(token|password|secret)\b)`
- **Severity:** critical
- **Description:** Detects sensitive data stored in NSUserDefaults/UserDefaults, which is a plaintext plist file readable on jailbroken devices and included in unencrypted device backups.
- **FP Indicators:**
  - Storing non-sensitive preferences (theme, locale, feature flags)
  - Value is encrypted before storage
  - Located in test/mock files
  - Reading a boolean flag like `hasCompletedAuth`
- **Fix:** Use Keychain Services for sensitive data: `SecItemAdd` / `SecItemCopyMatching`. Use a wrapper like `KeychainAccess` or `SwiftKeychainWrapper`. NSUserDefaults should only store non-sensitive preferences.

---

## 2. Keychain Access Control Missing

- **Name:** ios-keychain-access-control
- **Regex:** `(SecItemAdd\s*\((?![\s\S]*kSecAttrAccessControl|[\s\S]*kSecAttrAccessible)|kSecAttrAccessibleAlways\b|kSecAttrAccessibleAlwaysThisDeviceOnly\b|SecAccessControlCreateWithFlags\s*\(.*0\s*\))`
- **Severity:** high
- **Description:** Detects Keychain items stored without proper access control flags, making them accessible when the device is locked, or using deprecated accessibility values.
- **FP Indicators:**
  - Access control is set in a wrapper function
  - Using `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` (correct)
  - Located in a keychain utility class that sets flags centrally
- **Fix:** Set proper Keychain accessibility: `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`. Add biometric protection with `SecAccessControlCreateWithFlags(nil, .privateKeyUsage, .biometryCurrentSet, &error)`. Never use `kSecAttrAccessibleAlways`.

---

## 3. App Transport Security Disabled

- **Name:** ios-ats-disabled
- **Regex:** `(NSAppTransportSecurity.*NSAllowsArbitraryLoads\s*.*true|NSAllowsArbitraryLoads\s*</key>\s*<true/>|NSExceptionAllowsInsecureHTTPLoads\s*.*true|NSTemporaryExceptionAllowsInsecureHTTPLoads)`
- **Severity:** high
- **Description:** Detects App Transport Security exceptions that allow insecure HTTP connections, bypassing iOS's built-in network security.
- **FP Indicators:**
  - Exception scoped to specific development/test domains only
  - Required for legacy third-party SDK integration
  - Temporary exception with documented plan to remove
  - Local network discovery (Bonjour)
- **Fix:** Remove `NSAllowsArbitraryLoads`. Add domain-specific exceptions only when absolutely needed: `NSExceptionDomains` with `NSExceptionMinimumTLSVersion` set to `TLSv1.2`. File Apple exemption request if needed.

---

## 4. UIWebView Usage

- **Name:** ios-uiwebview-deprecated
- **Regex:** `(UIWebView\b|import\s+.*UIWebView|@IBOutlet.*UIWebView|\[\s*UIWebView\s|class.*:\s*UIWebView)`
- **Severity:** high
- **Description:** Detects usage of UIWebView which is deprecated, has known security vulnerabilities, and Apple rejects new submissions containing it since December 2020.
- **FP Indicators:**
  - Located in legacy code being migrated
  - Referenced in documentation or comments only
  - Third-party SDK dependency (not direct usage)
- **Fix:** Migrate to WKWebView which has better security, performance, and works in a separate process. Replace `UIWebView` with `WKWebView` and update delegate methods to `WKNavigationDelegate`.

---

## 5. NSURLSession Insecure Configuration

- **Name:** ios-nsurlsession-insecure
- **Regex:** `(URLSession.*\.allowsCellularAccess|didReceive.*challenge.*completionHandler\s*\(\s*\.useCredential|\.serverTrust\s*!\s*=\s*nil\s*\{.*completionHandler\s*\(\.useCredential|didReceive.*URLAuthenticationChallenge.*\.performDefaultHandling|allowsConstrainedNetworkAccess\s*=\s*false.*\.ephemeral)`
- **Severity:** high
- **Description:** Detects NSURLSession configurations that blindly accept server certificates or bypass SSL verification, enabling man-in-the-middle attacks.
- **FP Indicators:**
  - Proper certificate pinning is implemented in the delegate
  - Challenge handler validates the certificate chain
  - Test code or development-only network configuration
- **Fix:** Implement proper certificate validation in `urlSession(_:didReceive:completionHandler:)`. Use certificate or public key pinning. Never blindly call `completionHandler(.useCredential)` without validation.

---

## 6. Jailbreak Detection Missing

- **Name:** ios-jailbreak-detection
- **Regex:** `(isJailbroken|canOpenURL.*cydia|fileExists.*\/Applications\/Cydia|fileExists.*\/Library\/MobileSubstrate|DTTJailbreakDetection|IOSSecuritySuite|freeRASP)`
- **Severity:** medium
- **Description:** Checks for jailbreak detection implementation. Absence means the app runs on compromised devices without any security posture check.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - Low-risk apps may not need jailbreak detection
  - Detection is handled by an MDM/EMM solution
  - App deliberately supports jailbroken devices
- **Fix:** Implement jailbreak detection using multiple checks: Cydia URL scheme, suspicious file paths, sandbox integrity, writable system paths. Use `IOSSecuritySuite` for comprehensive detection. Decide on policy: warn, restrict features, or block.

---

## 7. UIPasteboard Sensitive Data

- **Name:** ios-uipasteboard-sensitive
- **Regex:** `(UIPasteboard\.general\.(string|setValue|setItems|setData)\s*=?\s*.*\b(token|password|secret|key|credential|ssn|credit)\b|UIPasteboard\.general\.string\s*=|\.copy\s*\(.*\b(token|password|secret)\b)`
- **Severity:** high
- **Description:** Detects sensitive data being copied to the system pasteboard, which is accessible to all apps and may sync via Universal Clipboard to other devices.
- **FP Indicators:**
  - Copying non-sensitive data (display name, reference ID)
  - Using a private pasteboard with `withUniqueName()`
  - Pasteboard is set with `localOnly: true` and `expirationDate`
- **Fix:** Never copy sensitive data to pasteboard. If necessary, use a local pasteboard with expiration: `UIPasteboard.general.setItems([[key: value]], options: [.localOnly: true, .expirationDate: Date(timeIntervalSinceNow: 60)])`. Clear pasteboard when app backgrounds.

---

## 8. NSLog Sensitive Data

- **Name:** ios-nslog-sensitive
- **Regex:** `(NSLog\s*\(\s*@?["\'].*\b(token|password|secret|key|credential|auth|session|jwt|bearer)\b|os_log\s*\(.*\b(token|password|secret)\b|print\s*\(\s*["\'].*\b(token|password|secret|key)\b.*\b(value|data|=)\b)`
- **Severity:** high
- **Description:** Detects NSLog/print/os_log statements containing sensitive data. NSLog output is accessible via Console.app, device logs, and crash reports.
- **FP Indicators:**
  - Logging the word but not the value (e.g., "token refreshed")
  - Inside `#if DEBUG` conditional compilation
  - Using os_log with `.private` privacy level
  - Test code with mock values
- **Fix:** Remove all sensitive data logging. Use `os_log` with `.private` for sensitive values: `os_log("Token: %{private}@", token)`. Wrap debug logs in `#if DEBUG`. Use OSLog subsystem with appropriate log levels.

---

## 9. Insecure Data Protection

- **Name:** ios-data-protection
- **Regex:** `(FileProtectionType\.none|\.noFileProtection|NSFileProtectionNone|NSDataWritingFileProtectionNone|\.completeUntilFirstUserAuthentication(?!.*override))`
- **Severity:** high
- **Description:** Detects files created without data protection, making them accessible even when the device is locked or via unencrypted backups.
- **FP Indicators:**
  - Files that need background access (e.g., background downloads)
  - Non-sensitive cached data or temp files
  - Located in app extension that needs before-first-unlock access
- **Fix:** Set file protection to `.complete` for sensitive files: `try data.write(to: url, options: .completeFileProtection)`. Set default protection in entitlements: `NSFileProtectionComplete`. Use `.completeUnlessOpen` only when background write access is needed.

---

## 10. Insecure Biometric Implementation

- **Name:** ios-biometric-insecure
- **Regex:** `(LAContext\s*\(\)\.evaluatePolicy\s*\(\.deviceOwnerAuthentication(?!BiometricsWith)|\.evaluatePolicy\s*\(\s*\.deviceOwnerAuthentication\s*,(?!.*biometry)|\.(canEvaluatePolicy|evaluatePolicy)\s*\(.*\)\s*\{[^}]*success\s*(==|!=)\s*(true|false)[^}]*\})`
- **Severity:** high
- **Description:** Detects biometric authentication implementations that may be bypassable: using device passcode fallback instead of biometrics-only, or weak success/failure handling.
- **FP Indicators:**
  - Intentional fallback to device passcode for accessibility
  - Success triggers Keychain access (proper security binding)
  - Located in test code
- **Fix:** Use `.deviceOwnerAuthenticationWithBiometrics` to require biometric auth. Bind biometric auth to Keychain access control: store credentials in Keychain with `.biometryCurrentSet` flag. Verify authentication result securely on the server side.
