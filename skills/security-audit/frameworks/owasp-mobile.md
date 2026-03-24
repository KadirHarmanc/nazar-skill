# OWASP Mobile Top 10 (2024) - Mobile Application Security

Reference framework for deep security audit of mobile applications (iOS/Android/React Native/Flutter).

---

## M1 - Improper Credential Usage

**CWE References:** CWE-200, CWE-256, CWE-312, CWE-522, CWE-798

**Grep Patterns:**
```
# Hardcoded API keys
pattern: "(api_key|apiKey|API_KEY|api_secret|apiSecret)\s*[:=]\s*['\"][A-Za-z0-9_\-]{16,}"
context: "Hardcoded API key in mobile code - shipped to device"

# Hardcoded credentials
pattern: "(password|secret|token|private_key|auth_token)\s*[:=]\s*['\"][^'\"]{8,}"
context: "Hardcoded credential in source code"

# Firebase/AWS/GCP keys
pattern: "(AIza[0-9A-Za-z_-]{35}|AKIA[0-9A-Z]{16}|sk-[a-zA-Z0-9]{32,})"
context: "Cloud provider API key"

# Credentials in SharedPreferences/UserDefaults
pattern: "(SharedPreferences|UserDefaults|NSUserDefaults|getSharedPreferences).*?(password|token|secret|key)"
context: "Credentials stored in insecure local storage"

# Keychain/Keystore not used
pattern: "(localStorage|AsyncStorage|SecureStore|MMKV).*?(password|token|secret|credential)"
context: "Check if Keychain/Keystore is used instead"

# Biometric without binding
pattern: "(BiometricPrompt|LAContext|TouchID|FaceID|biometric)"
context: "Check if biometric is bound to cryptographic key"

# Token in URL
pattern: "(token|key|secret|auth)\s*=.*?[?&]"
context: "Sensitive credential in URL parameter"
```

**Context Analysis Instructions:**
1. Search for any hardcoded secrets, API keys, or credentials in the codebase.
2. Verify credentials are stored in platform secure storage (iOS Keychain, Android Keystore).
3. Check that biometric authentication is bound to a cryptographic key, not just a boolean gate.
4. Verify API keys are not embedded in the app bundle (they can be extracted).
5. Check that tokens are not passed in URL query parameters (logged in server/proxy logs).
6. Verify certificate pinning is implemented for credential transmission.

**Example Exploit Scenario:**
A React Native app stores the user's auth token in AsyncStorage (unencrypted). An attacker with physical access or a backup extraction tool reads the token from the device's local storage. The app also has a hardcoded Firebase API key in the JavaScript bundle, which the attacker extracts and uses to access the Firebase project directly.

**Fix Guide:**
- Never hardcode credentials in mobile code; use server-side credential management.
- Store tokens in iOS Keychain or Android Keystore.
- For React Native, use `react-native-keychain` or `expo-secure-store` instead of AsyncStorage.
- Bind biometric authentication to a cryptographic key operation, not a boolean flag.
- Implement certificate pinning for all API communication.
- Use backend proxy for third-party API calls that require secrets.

---

## M2 - Inadequate Supply Chain Security

**CWE References:** CWE-426, CWE-494, CWE-829, CWE-830

**Grep Patterns:**
```
# Unpinned dependencies
pattern: "\"[^\"]+\":\s*\"[\^~>*]|>=\s*\d"
context: "Unpinned dependency version - supply chain risk"

# No lock file
pattern: "package-lock\.json|yarn\.lock|Podfile\.lock|pubspec\.lock"
context: "Check if lock file exists and is committed"

# Third-party SDKs
pattern: "(analytics|firebase|facebook|google|appsflyer|adjust|amplitude|mixpanel|sentry|crashlytics)"
context: "Third-party SDK - verify trust and necessity"

# Native library loading
pattern: "(System\.loadLibrary|dlopen|NSBundle.*load|dlsym)"
context: "Dynamic library loading - verify source"

# Custom plugin/module sources
pattern: "(git\+https|git\+ssh|github\.com.*\.git|pod.*:git)"
context: "Dependency from git source - verify integrity"

# Build script execution
pattern: "(postinstall|preinstall|prebuild|postbuild)\s*:"
context: "Package lifecycle script - possible malicious execution"
```

**Context Analysis Instructions:**
1. Verify all dependency lock files exist and are committed to version control.
2. Check for unpinned dependency versions that could be hijacked.
3. Review third-party SDKs for necessity and data collection practices.
4. Check for postinstall/preinstall scripts in dependencies.
5. Verify dependencies are from official registries, not arbitrary git repos.
6. Check if SBOM (Software Bill of Materials) is generated.

**Example Exploit Scenario:**
A mobile app uses unpinned lodash dependency. A malicious actor publishes a compromised version. The next install pulls the compromised version, which exfiltrates environment variables during the build process.

**Fix Guide:**
- Pin all dependencies to exact versions.
- Commit lock files and verify them in CI.
- Run dependency scanning in CI/CD pipeline.
- Review third-party SDKs quarterly; remove unnecessary ones.
- Generate and maintain SBOM.
- Configure code signing for all release builds.

---

## M3 - Insecure Authentication/Authorization

**CWE References:** CWE-287, CWE-306, CWE-307, CWE-862, CWE-863

**Grep Patterns:**
```
# Client-side auth checks
pattern: "(isAdmin|isLoggedIn|isAuthenticated|userRole|hasPermission)\s*=.*?(AsyncStorage|UserDefaults|SharedPreferences|localStorage)"
context: "Authorization decision based on client-side storage"

# Missing server-side validation
pattern: "(if\s*\(.*?(role|admin|permission).*?\).*?(navigate|push|render|show))"
context: "Client-side role check without server enforcement"

# Token validation
pattern: "(Bearer|Authorization|x-auth-token|x-access-token)"
context: "Check token validation on every API call"

# Session management
pattern: "(session|token).*?(expire|expiry|ttl|timeout|maxAge)"
context: "Check session/token expiration configuration"

# Deep link auth bypass
pattern: "(deeplink|universal.*?link|app.*?link|scheme.*?://)"
context: "Check if deep links bypass authentication"
```

**Context Analysis Instructions:**
1. Verify all authorization decisions are made server-side, not just in the mobile client.
2. Check that every API endpoint validates the auth token.
3. Verify biometric authentication cannot be bypassed by modifying the app binary.
4. Check session/token expiration is enforced.
5. Verify deep links cannot bypass authentication flows.

**Example Exploit Scenario:**
A banking app checks `if (userRole === 'admin')` in the React Native code to show admin features. An attacker uses Frida to hook the role check function and return true, gaining access to admin functionality. The server API does not validate the user's role.

**Fix Guide:**
- All authorization checks must happen server-side; client-side checks are only for UX.
- Validate auth token on every API request with server-side middleware.
- Use short-lived access tokens (15 min) with refresh token rotation.
- Protect deep links: always validate authentication state before rendering protected content.

---

## M4 - Insufficient Input/Output Validation

**CWE References:** CWE-20, CWE-74, CWE-79, CWE-89, CWE-471, CWE-915

**Grep Patterns:**
```
# WebView JavaScript injection
pattern: "(evaluateJavaScript|loadUrl.*javascript|postMessage|onMessage)"
context: "WebView JavaScript bridge - check input validation"

# SQL injection in local DB
pattern: "(rawQuery|execSQL|sqlite3_exec|executeSql).*?[\+\$`]"
context: "Raw SQL with string concatenation in local database"

# Intent/deeplink data usage
pattern: "(getIntent|getStringExtra|getData|intent\.data|launchOptions|userActivity)"
context: "Data from intent/deeplink used without validation"

# Path traversal
pattern: "(openFile|readFile|writeFile|FileManager).*?(request|intent|params|query)"
context: "File operation with external input"

# Clipboard data
pattern: "(UIPasteboard|ClipboardManager|Clipboard\.getString|getClipboardData)"
context: "Data from clipboard used without validation"
```

**Context Analysis Instructions:**
1. Check all WebView JavaScript bridges for input validation.
2. Verify local database queries use parameterized statements.
3. Check that data from intents, deep links, and external sources is validated before use.
4. Verify file operations validate paths (no path traversal).

**Example Exploit Scenario:**
A mobile app has a deep link handler where the userId is passed directly to a local SQLite query via string concatenation. An attacker crafts a malicious link with SQL injection payload and sends it via SMS. When the victim clicks the link, their auth tokens are exposed through UNION-based extraction.

**Fix Guide:**
- Use parameterized queries for local databases.
- Validate and sanitize all data from intents, deep links, and IPC.
- Use allowlists for deep link parameters (expected format, length, character set).
- Validate file paths against a base directory to prevent traversal.

---

## M5 - Insecure Communication

**CWE References:** CWE-295, CWE-296, CWE-297, CWE-298, CWE-319, CWE-757

**Grep Patterns:**
```
# HTTP instead of HTTPS
pattern: "http://(?!localhost|127\.0\.0\.1|10\.|192\.168)"
context: "Plaintext HTTP communication"

# Certificate pinning disabled
pattern: "(SSLPinning.*false|TrustAll|AllowAll|ALLOW_ALL_HOSTNAME|trustAllCerts|disable.*ssl|ssl.*verify.*false)"
context: "SSL/TLS validation disabled"

# NSAllowsArbitraryLoads
pattern: "(NSAllowsArbitraryLoads|NSExceptionAllowsInsecureHTTPLoads|NSExceptionMinimumTLSVersion)"
context: "iOS ATS exception allowing insecure connections"

# Android cleartext traffic
pattern: "(cleartextTrafficPermitted|usesCleartextTraffic|android:usesCleartextTraffic)"
context: "Android cleartext traffic allowed"

# Missing certificate pinning
pattern: "(OkHttp|URLSession|NSURLSession|HttpClient|axios|fetch)"
context: "HTTP client - check if certificate pinning is configured"

# WebSocket without TLS
pattern: "ws://(?!localhost|127\.0\.0\.1)"
context: "Unencrypted WebSocket connection"
```

**Context Analysis Instructions:**
1. Verify all network communication uses HTTPS/TLS.
2. Check for certificate pinning implementation.
3. Verify iOS ATS does not have broad exceptions.
4. Check Android Network Security Config does not allow cleartext traffic.
5. Verify custom TrustManagers do not blindly accept all certificates.

**Example Exploit Scenario:**
A mobile banking app sets NSAllowsArbitraryLoads: true in Info.plist for development and ships it to production. An attacker on the same WiFi performs a MITM attack, intercepts all API traffic, and captures auth tokens and banking credentials.

**Fix Guide:**
- Enforce HTTPS for all connections; no exceptions for production.
- Implement certificate pinning (TrustKit for iOS, OkHttp CertificatePinner for Android).
- Remove NSAllowsArbitraryLoads from Info.plist.
- Set `android:usesCleartextTraffic="false"` in AndroidManifest.xml.

---

## M6 - Inadequate Privacy Controls

**CWE References:** CWE-200, CWE-359, CWE-532, CWE-538, CWE-862

**Grep Patterns:**
```
# PII collection
pattern: "(email|phone|name|address|birthdate|ssn|national_id|location|gps|latitude|longitude)"
context: "PII data collection - verify purpose and consent"

# Tracking/analytics
pattern: "(analytics|tracking|fingerprint|deviceId|advertisingId|IDFA|GAID|IDFV)"
context: "User tracking - verify consent and disclosure"

# Location tracking
pattern: "(CLLocationManager|LocationManager|geolocation|getCurrentPosition|watchPosition|ACCESS_FINE_LOCATION)"
context: "Location data collection - verify minimization"

# Camera/microphone
pattern: "(AVCaptureSession|camera|microphone|MediaRecorder|getUserMedia)"
context: "Camera/microphone access - verify purpose limitation"

# Log PII
pattern: "(console\.(log|info|warn|error)|NSLog|Log\.(d|i|w|e)|print)\s*\(.*?(email|phone|name|user|password|token)"
context: "PII in debug logs - shipped to production"

# Clipboard PII
pattern: "(UIPasteboard|ClipboardManager|Clipboard\.setString|copy.*?(password|token|card|ssn))"
context: "Sensitive data copied to clipboard"
```

**Context Analysis Instructions:**
1. Identify all PII collected and verify each has a legitimate purpose.
2. Check that privacy consent is obtained before data collection.
3. Verify analytics/tracking respects user opt-out preferences.
4. Check that PII is not logged in production builds.
5. Verify sensitive screens prevent screenshots.

**Example Exploit Scenario:**
A fitness app collects precise GPS location every 30 seconds in the background. The privacy policy does not disclose background location tracking. Debug logs include full user profiles with email, phone, and health data, accessible on older Android devices.

**Fix Guide:**
- Implement data minimization: only collect necessary data with user consent.
- Remove PII from all log statements in production builds.
- Provide opt-out for analytics and tracking.
- Conduct a privacy impact assessment.

---

## M7 - Insufficient Binary Protections

**CWE References:** CWE-284, CWE-693, CWE-919

**Grep Patterns:**
```
# Debugging enabled
pattern: "(android:debuggable|DEBUG|isDebuggerConnected|sysctl|ptrace)"
context: "Debug configuration - verify disabled in release"

# Anti-tampering missing
pattern: "(checkSignature|verifyIntegrity|SafetyNet|DeviceCheck|AppAttest|Play Integrity)"
context: "Check if app integrity verification is implemented"

# Jailbreak/root detection
pattern: "(isJailbroken|isRooted|checkRoot|detectRoot|canOpenURL.*cydia|su.*binary)"
context: "Check if jailbreak/root detection is implemented"

# Code obfuscation
pattern: "(ProGuard|R8|obfusc|minifyEnabled|hermes)"
context: "Check if code obfuscation is configured"

# Sensitive logic in client
pattern: "(validateLicense|checkSubscription|isPremium|isTrialExpired|verifyPurchase)"
context: "Business logic that should be server-validated"
```

**Context Analysis Instructions:**
1. Check if the release build has debug mode disabled.
2. Verify code obfuscation is configured.
3. Check for jailbreak/root detection implementation.
4. Verify sensitive business logic is server-side, not only in the client.

**Example Exploit Scenario:**
An attacker decompiles the Android APK (no ProGuard) and finds the subscription validation logic. They patch the code to always return true, repackage, and sign the APK. No integrity check detects the modification.

**Fix Guide:**
- Enable ProGuard/R8 with strict rules for Android release builds.
- Implement runtime integrity checks: SafetyNet/Play Integrity (Android), DeviceCheck/AppAttest (iOS).
- Move business logic (subscription, licensing) to server-side validation.

---

## M8 - Security Misconfiguration

**CWE References:** CWE-16, CWE-276, CWE-316, CWE-489, CWE-732

**Grep Patterns:**
```
# Exported components
pattern: "(exported\s*=\s*true|android:exported|intent-filter)"
context: "Exported Android component - check necessity"

# Backup enabled
pattern: "(android:allowBackup|android:fullBackupContent)"
context: "App backup may expose sensitive data"

# WebView misconfiguration
pattern: "(setJavaScriptEnabled|javaScriptEnabled|setAllowFileAccess|allowFileAccess)"
context: "WebView security configuration"

# Insecure file permissions
pattern: "(MODE_WORLD_READABLE|MODE_WORLD_WRITABLE|0666|0777|chmod\s*777)"
context: "Insecure file permissions"

# Excessive permissions
pattern: "(ACCESS_FINE_LOCATION|READ_CONTACTS|READ_PHONE_STATE|CAMERA|RECORD_AUDIO|READ_SMS|ACCESS_BACKGROUND_LOCATION)"
context: "Sensitive permission - verify necessity"
```

**Context Analysis Instructions:**
1. Check Android Manifest for exported components, permissions, and backup settings.
2. Check iOS Info.plist for ATS exceptions, URL schemes, and permissions.
3. Verify WebView configuration is secure.
4. Check file permissions are restrictive.
5. Verify only necessary permissions are requested.

**Example Exploit Scenario:**
An Android app has `android:allowBackup="true"` and an exported Activity. An attacker uses adb backup to extract the app's data directory containing an unencrypted SQLite database with user tokens.

**Fix Guide:**
- Set `android:allowBackup="false"`.
- Set `android:exported="false"` for components that do not need external access.
- Configure WebView securely: disable file access, restrict JavaScript.
- Request only necessary permissions.

---

## M9 - Insecure Data Storage

**CWE References:** CWE-312, CWE-313, CWE-316, CWE-532, CWE-922

**Grep Patterns:**
```
# Unencrypted local storage
pattern: "(SharedPreferences|UserDefaults|NSUserDefaults|AsyncStorage|localStorage|MMKV)"
context: "Local storage - check if sensitive data is encrypted"

# SQLite without encryption
pattern: "(SQLiteDatabase|sqlite3|realm|FMDB|expo-sqlite)"
context: "Local database - check if encrypted (SQLCipher)"

# External storage
pattern: "(EXTERNAL_STORAGE|getExternalFilesDir|getExternalStorageDirectory)"
context: "External storage usage - accessible by other apps"

# Cache sensitive data
pattern: "(cache|tmp|temp).*?(token|password|user|credential|session)"
context: "Sensitive data in cache/temp directory"

# Keyboard cache
pattern: "(autocorrect|autocomplete|textContentType|spellCheck|secureTextEntry)"
context: "Keyboard may cache sensitive input"
```

**Context Analysis Instructions:**
1. Check what data is stored locally and whether encryption is applied.
2. Verify sensitive data uses platform secure storage (Keychain, Keystore).
3. Check SQLite databases for encryption (SQLCipher).
4. Verify sensitive data is not stored on external storage.
5. Check that keyboard caching is disabled for sensitive input fields.

**Example Exploit Scenario:**
A healthcare app stores patient records in a plain SQLite database. On a rooted device, any app with root access can read the database. The auth token is stored in SharedPreferences (unencrypted).

**Fix Guide:**
- Use EncryptedSharedPreferences (Android) or Keychain (iOS) for sensitive data.
- Encrypt local databases with SQLCipher.
- Never store sensitive data on external storage.
- Disable autocorrect on sensitive input fields: `secureTextEntry={true}`.

---

## M10 - Insufficient Cryptography

**CWE References:** CWE-261, CWE-326, CWE-327, CWE-328, CWE-330, CWE-338

**Grep Patterns:**
```
# Weak algorithms
pattern: "(MD5|SHA1|DES|RC4|ECB|Blowfish|ROT13|base64.*encrypt)"
context: "Weak or deprecated cryptographic algorithm"

# Insecure random
pattern: "(Math\.random|random\.random|Random\(\)|java\.util\.Random)"
context: "Insecure random number generator for security purposes"

# Hardcoded IV/key
pattern: "(iv|IV|nonce|key)\s*[:=]\s*['\"][A-Fa-f0-9]{16,}"
context: "Hardcoded cryptographic key or IV"

# ECB mode
pattern: "(ECB|AES/ECB|createCipheriv.*ecb)"
context: "ECB mode produces identical ciphertext for identical plaintext"

# Base64 used as encryption
pattern: "(btoa|atob|base64.*encode|base64.*decode).*?(password|secret|token|encrypt)"
context: "Base64 encoding misused as encryption - not actual encryption"

# Deprecated crypto APIs
pattern: "(crypto\.createCipher\b|Cipher\.getInstance\s*\(\s*['\"]AES['\"]|CCCrypt)"
context: "Deprecated crypto API without explicit mode/padding"
```

**Context Analysis Instructions:**
1. Verify all cryptographic operations use strong algorithms (AES-256-GCM, ChaCha20, SHA-256+).
2. Check that cryptographic random is used for security-sensitive values.
3. Verify encryption keys are not hardcoded and are properly managed.
4. Check that encryption uses appropriate modes (GCM, not ECB).
5. Verify base64 is not used as a substitute for encryption.

**Example Exploit Scenario:**
A messaging app uses AES-ECB mode for message encryption. Because ECB produces identical ciphertext blocks for identical plaintext blocks, an attacker can detect patterns. The encryption key is derived from the user's password using MD5 without salt, making it trivially crackable.

**Fix Guide:**
- Use AES-256-GCM (authenticated encryption) instead of ECB.
- Use platform-native cryptographic random for IVs and nonces.
- Never hardcode encryption keys; use Android Keystore or iOS Keychain.
- Use PBKDF2 (100,000+ iterations), scrypt, or Argon2 for key derivation from passwords.
- Never use base64 as encryption; it is encoding, not encryption.
