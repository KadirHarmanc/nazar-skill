# Android Native Security Rules

Security patterns for Kotlin and Java Android applications.

---

## 1. SharedPreferences MODE_WORLD_READABLE

- **Name:** android-sharedprefs-world-readable
- **Regex:** `(MODE_WORLD_READABLE|MODE_WORLD_WRITEABLE|getSharedPreferences\s*\(\s*["\'][^"\']+["\']\s*,\s*(1|2|Context\.MODE_WORLD)|SharedPreferences.*\b(token|secret|password|key|credential|auth|session|jwt)\b|\.edit\s*\(\s*\)\.put(String|Int|Boolean)\s*\(\s*["\'].*(token|password|secret|key|credential))`
- **Severity:** critical
- **Description:** Detects SharedPreferences used with world-readable/writable modes (deprecated and insecure), or sensitive data stored in SharedPreferences which is a plaintext XML file accessible on rooted devices.
- **FP Indicators:**
  - MODE_PRIVATE is used (correct mode)
  - Storing non-sensitive preferences (theme, locale)
  - Value is encrypted before storage (EncryptedSharedPreferences)
  - Located in test/mock code
- **Fix:** Always use `MODE_PRIVATE`. Use `EncryptedSharedPreferences` from AndroidX Security for sensitive data: `EncryptedSharedPreferences.create(context, "secret_prefs", masterKey, ...)`. For highly sensitive data, use Android Keystore.

---

## 2. WebView JavaScript Enabled

- **Name:** android-webview-js-enabled
- **Regex:** `(setJavaScriptEnabled\s*\(\s*true\s*\)|\.settings\.javaScriptEnabled\s*=\s*true|addJavascriptInterface\s*\(|setAllowFileAccessFromFileURLs\s*\(\s*true\s*\)|setAllowUniversalAccessFromFileURLs\s*\(\s*true\s*\)|setAllowFileAccess\s*\(\s*true\s*\))`
- **Severity:** high
- **Description:** Detects WebView configurations that enable JavaScript execution, JS-to-native bridges, or file access, which can lead to XSS, arbitrary code execution, or local file theft.
- **FP Indicators:**
  - WebView loads only trusted first-party HTTPS content
  - JavaScript interface methods are annotated with `@JavascriptInterface` (required API 17+)
  - File access is disabled and content loads from network only
  - Located in a controlled in-app browser
- **Fix:** Disable JavaScript if not needed. If JavaScript is required, load only HTTPS content from trusted domains. Remove `addJavascriptInterface` on API < 17. Set `setAllowFileAccessFromFileURLs(false)` and `setAllowUniversalAccessFromFileURLs(false)`. Validate all URLs in `shouldOverrideUrlLoading`.

---

## 3. Intent Validation Missing

- **Name:** android-intent-validation
- **Regex:** `(getIntent\s*\(\s*\)\.(getStringExtra|getIntExtra|getData|getExtras|getBundleExtra)\s*\((?![\s\S]{0,100}(validate|check|verify|sanitize|require))|intent\.(getStringExtra|getData|getExtras)\s*\(.*(?!.*\b(if|when|require|check)\b))`
- **Severity:** high
- **Description:** Detects Intent data extraction without apparent validation. Intents from other apps can contain malicious data that leads to injection or logic bypass.
- **FP Indicators:**
  - Validation happens in a separate method after extraction
  - Intent is from a trusted internal component (explicit intent)
  - Data is type-safe (getIntExtra with default value)
  - Located inside a validation block
- **Fix:** Validate all Intent extras before use. Check for null/empty values. Validate format and range. For URI data, verify scheme and host: `intent.data?.let { uri -> require(uri.scheme == "https" && uri.host == "myapp.com") }`.

---

## 4. Exported Components Unprotected

- **Name:** android-exported-components
- **Regex:** `(android:exported\s*=\s*["\']true["\'](?![\s\S]*android:permission)|<activity(?![\s\S]*android:exported\s*=\s*["\']false["\'])[\s\S]*<intent-filter|<receiver(?![\s\S]*android:permission)[\s\S]*android:exported\s*=\s*["\']true|<provider(?![\s\S]*android:permission)[\s\S]*android:exported\s*=\s*["\']true)`
- **Severity:** high
- **Description:** Detects activities, receivers, services, and content providers exported without permission protection, allowing any app to interact with them.
- **FP Indicators:**
  - Component is intentionally public (launcher activity, share target)
  - Permission is defined at application level
  - Located in AndroidManifest for a library with documented public API
  - Main/launcher activity (must be exported)
- **Fix:** Set `android:exported="false"` for internal components. For components that must be exported, add permission protection: `android:permission="com.myapp.permission.MY_PERMISSION"`. Use signature-level permissions for same-developer apps.

---

## 5. Broadcast Receiver Security

- **Name:** android-broadcast-receiver
- **Regex:** `(registerReceiver\s*\(\s*\w+\s*,\s*\w+(?!\s*,\s*(RECEIVER_NOT_EXPORTED|Context\.RECEIVER_NOT_EXPORTED))|sendBroadcast\s*\(\s*\w+(?!\s*,\s*["\'])|LocalBroadcastManager(?!.*\.getInstance)|registerReceiver\s*\(\s*\w+\s*,\s*\w+\s*,\s*null)`
- **Severity:** high
- **Description:** Detects broadcast receivers registered without export control or broadcasts sent without permission restrictions, allowing interception or spoofing by other apps.
- **FP Indicators:**
  - Using `RECEIVER_NOT_EXPORTED` flag (API 33+, correct)
  - Using `LocalBroadcastManager` for internal broadcasts (correct)
  - Broadcast has permission parameter set
  - System broadcast receiver (BOOT_COMPLETED, BATTERY_LOW)
- **Fix:** Use `registerReceiver(receiver, filter, RECEIVER_NOT_EXPORTED)` for internal receivers (API 33+). Use `LocalBroadcastManager` or explicit intents for internal communication. Add permission parameter to `sendBroadcast`. For Android 14+, specify export behavior.

---

## 6. Certificate Pinning Missing

- **Name:** android-cert-pinning
- **Regex:** `(CertificatePinner|network_security_config|NetworkSecurityPolicy|TrustManagerFactory|X509TrustManager|ssl-pinning|okhttp3.*\.certificatePinner)`
- **Severity:** medium
- **Description:** Checks for certificate pinning implementation. Absence allows man-in-the-middle attacks with rogue CA certificates installed on the device.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - Pinning is configured in `network_security_config.xml`
  - Using OkHttp CertificatePinner (correct approach)
  - Backend uses short-lived certificates with frequent rotation
- **Fix:** Implement certificate pinning using `network_security_config.xml` with pin-set entries, or OkHttp `CertificatePinner`. Pin to intermediate CA or leaf certificate. Include backup pins for rotation: `<pin-set><pin digest="SHA-256">base64==</pin><pin digest="SHA-256">backup-base64==</pin></pin-set>`.

---

## 7. Root Detection Missing

- **Name:** android-root-detection
- **Regex:** `(RootBeer|isDeviceRooted|checkForRoot|SafetyNet|PlayIntegrity|IntegrityManager|freeRASP|rootDetection|SuFile|SuperUser)`
- **Severity:** medium
- **Description:** Checks for root detection implementation. Absence means the app runs on rooted devices where all security assumptions (sandboxing, file permissions) are invalid.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - Low-risk apps may not need root detection
  - Detection is handled by MDM/EMM solution
  - App intentionally supports rooted devices
- **Fix:** Implement root detection using RootBeer library or Google Play Integrity API. Check for: su binary, Superuser.apk, writable system partition, test-keys build tags. Use `freeRASP` for comprehensive RASP protection.

---

## 8. Log.d Sensitive Data

- **Name:** android-log-sensitive
- **Regex:** `(Log\.(d|i|v|w|e)\s*\(\s*["\'][^"\']*["\'],\s*[^)]*\b(token|password|secret|key|credential|auth|session|jwt|bearer|cookie)\b|Timber\.(d|i|v|w|e)\s*\(\s*["\'][^"\']*["\'].*\b(token|password|secret)\b|println\s*\(\s*["\'].*\b(token|password|secret)\b)`
- **Severity:** high
- **Description:** Detects Log.d/Timber/println statements containing sensitive data. Android logs are accessible via `adb logcat` by any app with READ_LOGS permission or USB debugging.
- **FP Indicators:**
  - Logging the word but not the value (e.g., `"Token refreshed successfully"`)
  - Inside `BuildConfig.DEBUG` conditional
  - Using Timber with release tree that strips logs
  - Test code with mock values
- **Fix:** Remove all sensitive data logging. Use Timber with a release tree that drops debug/verbose logs. Use ProGuard/R8 rules to strip Log calls: `-assumenosideeffects class android.util.Log { public static int d(...); }`. Never log full tokens or credentials.

---

## 9. rawQuery SQL Injection

- **Name:** android-rawquery-injection
- **Regex:** `(rawQuery\s*\(\s*["\'].*\+\s*\w|rawQuery\s*\(\s*["\'].*\$\{|rawQuery\s*\(\s*String\.format|execSQL\s*\(\s*["\'].*\+\s*\w|execSQL\s*\(\s*["\'].*\$\{|compileStatement\s*\(\s*["\'].*\+)`
- **Severity:** critical
- **Description:** Detects SQLite rawQuery/execSQL calls with string concatenation or interpolation, creating SQL injection vulnerabilities in local database operations.
- **FP Indicators:**
  - Using parameterized queries: `rawQuery("SELECT * WHERE id=?", arrayOf(id))`
  - Table/column names from code constants (not user input)
  - Using Room DAO with `@Query` annotation (safe)
  - Located in database migration with hardcoded values
- **Fix:** Use parameterized queries: `db.rawQuery("SELECT * FROM users WHERE id=?", arrayOf(userId))`. Use Room ORM with `@Query` annotations for compile-time verified queries. Use `ContentValues` for inserts/updates.

---

## 10. FileProvider Misconfiguration

- **Name:** android-fileprovider-misconfig
- **Regex:** `(android:authorities\s*=\s*["\'].*\.fileprovider["\'](?![\s\S]*android:exported\s*=\s*["\']false)|<external-path\s+name\s*=\s*["\'][^"\']*["\']\s+path\s*=\s*["\']\.?["\']|<root-path\s+name|grantUriPermission\s*=\s*["\']true["\'].*(?!android:exported\s*=\s*["\']false))`
- **Severity:** high
- **Description:** Detects FileProvider misconfigurations: exported provider, overly broad path sharing (root-path or external-path with "."), or permissive URI grants that expose app files.
- **FP Indicators:**
  - FileProvider is properly configured with `exported="false"`
  - Paths are scoped to specific directories
  - URI permissions are granted temporarily via intent flags
  - Located in a well-known library configuration
- **Fix:** Always set `android:exported="false"` for FileProvider. Use specific scoped paths instead of root-path: `<files-path name="images" path="images/" />`. Grant URI permissions via intent flags (FLAG_GRANT_READ_URI_PERMISSION) instead of manifest. Avoid `<root-path>` and broad `<external-path path=".">`.
